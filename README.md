# ORM: Preventing Record Duplication

## Objectives

1. Avoid creating duplicate records in a database that is mapped to a Ruby program 
2. Build a `#find_or_create_by` method

## The Dreaded Duplication

What happens when two Ruby objects get created using the same attributes? If we are trying to persist representations of such objects to a database, would we end up with essentially identical rows in our table? That would make for a very confusing database and our program would quickly become useless as a way to store and manage information. 

For example, lets say we have a `Song` class that produces individual song objects, each of which have a `name` and `album` attribute.

Nothing stops us from creating two objects, each of which have the exact same name and album. 

```ruby
hello = Song.new("Hello", "25")
hello_again = Song.new("Hello", "25")
```

What happens when we save these objects to our database?

*For this example, we'll assume our connection to the database is stored in `DB[:conn]`*

```ruby
hello.save
hello_again.save

DB[:conn].execute("SELECT * FROM songs WHERE name = "Hello")
# => [[1, "Hello", "25"], [2, "Hello", "25"]]
```

We have two records that contain the same information! How can we avoid this? When we try to save a new `Song` instance, we should first check to see the object we are trying to save already has an equivalent record in the database, if it does, we should simply update it, otherwise, we can go ahead and save it. 

## Saving vs. Updating

Let's say we have a song, `hello`:

```ruby
hello = Song.new("Hello", "25")
```

Before we call `#save` on our `hello` object, we need to check and see if a record containing this name and album already exists in the database. The SQL statement to accomplish that would look something like this:

```ruby
SELECT * FROM songs
WHERE name = "Hello", album = "25";
```

If this statement returns a record, we don't need to create a new record, only update the existing one. Otherwise, we need to insert a new record into our database table. 

Let's build a method that will allow us to either *find an existing record* or *create and save* a new one. 

## The `#find_or_create_by` Method

Take a look at our `Song` class. 

```ruby
class Song

attr_accessor :name, :album
attr_reader :id
  
  def initialize(id=nil, name, album)
    @id = id
    @name = name
    @album = album
  end

  def save
    if self.id 
      self.update
    else
      sql = <<-SQL
        INSERT INTO songs (name, album) 
        VALUES (?, ?)
      SQL

      DB[:conn].execute(sql, self.name, self.album)
      @id = DB[:conn].execute("SELECT last_insert_rowid() FROM students")[0][0]
    end
  end

  def self.create(name:, album:)
    student = Student.new(name, album)
    student.save
    student
  end
  
  def self.find_by_id(id)
    sql = "SELECT * FROM songs WHERE id = ?"
    result = DB[:conn].execute(sql, id)[0]
    Song.new(result[0], result[1], result[2])
  end
  
  def update
    sql = "UPDATE songs SET name = ?, album = ? WHERE id = ?"
    DB[:conn].execute(sql, self.name, self.album, self.id)
  end
end
``` 

Let's build our `#find_or_create_by` method:

```ruby
  def self.find_or_create_by(name:, album:)
    song = DB[:conn].execute("SELECT * FROM songs WHERE name = ? AND album = ?", name, album)
    if !song.empty?
      song_data = song[0]
      song = Song.new(song_data[0], song_data[1], song_data[2])
    else
      song = self.create(name: name, album: album)
    end
    song
  end 
```

Let's break this down: 

First, we query the database: does a record exist that has this name and album? 

```ruby
song = DB[:conn].execute("SELECT * FROM songs WHERE name = ? AND album = ?", name, album)
```

If such a record exists, the `song` variable will now point to an array that would look something like this:

*song name and album name provided as a hypothetical example*

```ruby
[[1, "Hello", "25"]]
```

If this is the case, then the statement: `!song.empty?` will return `true`. Therefore, we will *not* create a new song instance and record, we will instead grab and return the existing song record. That is what is accomplished with these lines:

```ruby
song_data = song[0]
song = Song.new(song_data[0], song_data[1], song_data[2])
```

We grab the `song_data` from the `song` array of arrays, setting `song_data` equal to:

```ruby
[1, "Hello", "25"]
```

Then, we use this array to create a new `Song` instance with the given id, name and album. 

However, if no record exists that matches the name and album passed in to this method as arguments, then `!song.empty?` will return false, and we will instead create and save a new `Song` instance with the `#create` method. 

At the end of our `#find_or_create_by` method, we will return the `song` object that we either found or created. 


### Our Code in Action 

Now, we can use our `Song` class without worrying about creating duplicate records:

```ruby
Song.find_or_create_by(name: "Hello", album: "25")
Song.find_or_create_by(name: "Hello", album: "25")

DB[:conn].execute("SELECT * FROM songs WHERE name = Hello, album = 25")
# => [[1, "Hello", "25"]]
```

Although we called `#find_or_create_by` twice *with the same data* (gasp!), we only created *one record with that data*. 

<a href='https://learn.co/lessons/orm-find-or-create-by' data-visibility='hidden'>View this lesson on Learn.co</a>

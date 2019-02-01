---
layout: post
title:      "Notable Books CLI Application"
date:       2019-02-01 11:46:38 -0500
permalink:  notable_books_cli_application
---


I’m feeling accomplished now that I’ve completed my Ruby CLI Data Gem project, which was my first experience in designing an application from start to finish. The project is a synthesis of everything we’ve learned so far: from procedural Ruby to object oriented programming; understanding and navigating the command line interface; and how to work with Git and GitHub. 

I learned so much from the process of planning and building this program and definitely experienced the whole spectrum of coding feelings from total panic when my application broke to the golden focus and confidence of getting it to work again. My constant reminder: **Break things down into small pieces. Figure out how to make just the next thing work. **

It took me quite a few attempts to find a website that was a good candidate for scraping. I kept running into sites that looked promising, but had content loaded with JavaScript. With the help of Jenn Hansen’s [Scraper Checker](https://repl.it/@jenn_leigh_hansen/ScraperChecker?language=ruby) on Repl.it, I settled on the New York Times’ [100 Notable Books of 2018](https://www.nytimes.com/interactive/2018/11/19/books/review/100-notable-books.html) list. 

I began by stubbing out what the command line interface (CLI) class would look like, and I decided that I wanted the user to have two main options for viewing book data:

* Browse books in numbered order (browsing all books, broken up into small numbered lists in order to make the command line output more readable.)
* Browse books sorted by Genre

For each option, I wanted the CLI to print a numbered list of books so the user could enter a number to see more information about their chosen book. The option to browse books by Genre would also require a way to keep track of which genre each book belonged to. 

From this planning, I decided on four classes for my program: 

* Scraper - Scrape data from the NYT site; instantiate new Books and Genres
* CLI - methods for execution of the application and user interaction
* Book - assign attributes to Books; make each Book belong to one or more genre(s)
* Genre - create Genre attributes; make each Genre have many books

To begin scraping, I found that all book data was nested under `div#g-book-data`. A book’s title and author were easy to scrape, as every book’s title could be accessed uniformly through `div#g-book-title` and the author through `div#g-book-author b`.

Everything else - as I would discover later in the coding process -  proved more difficult to scrape. Each book has 1-3 genres listed inside of the same div, formatted as a plain line of text. Every book has a price and a publisher, but the order of these details changes, and some books have additional info inside the div (translator name, book format, etc). Due to this lack of a uniform organization of some data points across the book list, I had to narrow my focus on which attributes to scrape. 

**Easy to scrape and parse:**
![](https://drive.google.com/uc?export=view&id=1g73trR_KnHR0dTRKKjADExfUxUqGTYjS)


**Not so easy:**
![](https://drive.google.com/uc?export=view&id=15W2IQH8IWOpI9VLCUZ5_TPCua18_NIt0)

Ultimately, I was able to scrape and use the data for a book’s Title, Author, Genre, Publisher, Price, and Description. Here’s what my scraper method became: 

```
  def self.scrape_book_info
    books_array = []
     scrape_page.css(".g-book-data").each.with_index do |nodeset|
       book_hash = {}

       details = nodeset.css(".g-book-author").text
        .split(". ").map!(&:strip)
        
       book_hash[:title] = nodeset.css(".g-book-title").text.strip
       book_hash[:author] = nodeset.css(".g-book-author b").text.strip.chomp(".")
       book_hash[:genre] = create_genres(nodeset.css(".g-book-tag").text
        .split(".").map!(&:strip).reject(&:empty?))
       book_hash[:description] = nodeset.css(".g-book-description").text.strip
       book_hash[:price] = details.sort.first
       book_hash[:publisher] = details.last.chomp(".")
       books_array << book_hash
      end

      NotableBooks2018::Book.create_from_collection(books_array)
  end
```

Inspired by the [Student Scraper](https://github.com/norawolf/oo-student-scraper-online-web-pt-112618) lab, I decided to iterate through the data for each book and populate a hash with the key/value pairs of attributes and data for each book. Inside of my Book class, I utilized metaprogramming within my initialize method to allow for me to easily alter which attributes each Book would be instantiated with. Following this format allowed me flexibility to include more Book attributes as I gradually worked out how to scrape some of the less accessible elements from the NYT page. 

```
  def initialize(book_hash)
    book_hash.each do |attribute, value|
      self.send("#{attribute}=", value)
    end
    @@all << self
  end
	
	  def self.create_from_collection(book_array)
    book_array.each do |book_hash|
      NotableBooks2018::Book.new(book_hash)
    end
  end
```

After scraping and instantiating my Books, it was time to move on to one of the most difficult aspects of this project: the object relationships between my Book and Genre classes. I took to my notebook to sketch things out:

<center>A Genre <b>has many</b> Books, and a Book <b>belongs to</b> one or several Genres.</center>

I altered my `scrape_book_info` method to also be responsible for instantiating new Genres while iterating through the scraped genre text data. At first, I ended up with 100 Genre objects... one for each book in my `Book.all` array. OOPS.

To fix this, I started with pseudocode:

*-This method should search the Genre.all array 
- and if the genre name being passed in matches the `.name` attribute for any Genre instance, 
- associate this book with that genre. 
- But if the genre doesn’t exist, it should create it. *

I puzzled for a while. Then: beautiful lightbulb moment! This was exactly the scenario for the `find_or_create_by` methods we practiced in the OO labs. Thus, inside my Genre class:

```
  def self.find_by_name(name)
    all.find {|genre| genre.name == name}
  end

  def self.create_by_name(name)
    NotableBooks2018::Genre.new(name)
  end
	
 def self.find_or_create_by_name(name)
    find_by_name(name) || create_by_name(name)
 end
```

And so, the Genre instantiation became a helper method inside of Scraper.rb that I called while assigning the value of my `book_hash[:genre]` attribute. Since Books have 1-3 genres, I iterated with `.collect` so that this method would return an array with either 1, 2, or 3 Genre instances.

```
def self.create_genres(css)
    genre_data = css
    genre_data.collect do |genre_name|
      NotableBooks2018::Genre.find_or_create_by_name(genre_name)
    end
end
```

Then, I asked the question: <b>How does a Genre know what books it has?</b>

The answer: It doesn’t! (Yet).

Over to the Book class, to create a custom genre= method, so that when I call .books on an instance of the Genre class, it returns actual Book instances instead of string names. 

  ```
	 def genre=(genre)
     @genre = genre
		
     #genre is an array of 1 or 2 or 3 instances
     # for each instance in the genre array (Fiction, Poetry, etc)
     # associate this Book instance being instantiated to that Genre
		
      genre.each do |genre_instance|
        genre_instance.books << self
      end
	end

	```

Now that my class relationships were working, I built out my CLI class with the many methods I would need to walk the user through the 2 main options for viewing books: by genre or from a numbered list. I put a lot of care into making the navigation flexible, allowing the user to switch between viewing books by Genre or index and allowing the user to exit anytime. I also tried to sanitize the data inputs and control for invalid data. 

I’m really, really proud of how this application turned out, and you can check it out for yourself over at Github: [https://github.com/norawolf/notable_books_2018](https://github.com/norawolf/notable_books_2018)

**Future Functionality Expansions**
I have many ideas for how I could continue to expand the functionality of this CLI program as I continue to learn more. Here are two that I would like to keep working on:

* Allow the user to create a profile to save books to a Reading List
* Add the ability for a user to check if a local library has a copy of a book from the list 

Thank you for reading, and I hope this gem helps you to expand your personal reading list for 2019!










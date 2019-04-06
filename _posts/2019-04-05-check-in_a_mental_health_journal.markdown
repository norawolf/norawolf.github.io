---
layout: post
title:      "Check-In: A Mental Health Journal"
date:       2019-04-05 22:35:19 -0400
permalink:  check-in_a_mental_health_journal
---

For my final Sinatra project, I created [Check-In: A Mental Health Journal](https://github.com/norawolf/check-in-journal). Check-In is an application designed to allow users to easily create entries to track their moods and associated daily activities. The further I get into my web development education, the more I am able to envision the links between the skills we’re learning and the software I would like to build to support health, accessibility, and community wellness. 

## Project Outline
I spent some time outlining the functionality I wanted my application to have:
* A user can create a new account
* A user can create a new entry
* An entry has both moods and activities, which are unique objects
* A user can edit an entry
* A user can delete an entry
* A user can see all of their own entries
* A user cannot see the entries of another user

I ended up with six models in my domain, two of which are join tables which allow instances of the Activity and Mood classes to be associated with multiple entries:
* User
* Entry
* Mood
* Activity
* ActivitiesEntry
* MoodsEntry

With the help of ActiveRecord macros, I outlined the following relationships:

```
Entry belongs to a User
Entry has many :moods_entries
	has many :moods, through: moods_entries
Entry has many :activities_entries
	has many :activities, through: activities_entries
Mood has many :moods_entries
	has many :entries, through: moods_entries
Activity has many :activities_entries
	has many :entries, through: activities_entries 
```

I installed the rails-erd gem to generate a diagram of my domain: 

![](http://drive.google.com/uc?export=view&id=1crHXoDlyMeAUc6chh8P82Ay4GfedWxZH)

I used the [corneal gem](https://github.com/thebrianemory/corneal) to generate my MVC project structure, and I then got to work stubbing out my routes. 

## Interesting Development Challenges
While building the form to create a new entry, I iterate through the collection of all activities, `Activity.all`, in order to generate a list of checkboxes with selectable activity names. When checking the `params` hash in my `post /entries/` route, I kept discovering that a mysterious new activity with the name of `on` was being created and added to Activity.all. Where was this `on` coming from?
  * As it turns out, when a user submits a form with a `checkbox` input, the form will default the value to `on` if no value attribute is specified . All I needed to do was add a value attribute to my `<input>` tag. Thanks, [StackOverflow](https://stackoverflow.com/a/13658228).
 
```
<% @activities.each do |activity| %>
    <input type="checkbox" class="label-body" name="entry[activities][]" value="<%= activity.name %>"> <%= activity.name %></input>
<% end %>
```

I ran into another issue of unexpected data appearing within my `params` hash after form submission. Inside of `post/entries/`, `params[:moods]` contained an empty string as the first value in the moods array. This resulted in my application instantiating a new mood object with an empty name value. 
   * My form uses the HTML `select multiple` tag in order to allow a user to select multiple moods from a dropdown list to add to their entry. According to [API Dock](https://apidock.com/rails/ActionView/Helpers/FormOptionsHelper/select), when a multiple parameter is passed to select, an auxillary hidden field is generated before every multiple select. 
   * This is an HTML workaround to address the fact that when all options get deselected from a multiple select, web browsers do not send any value to the server. 
   * So, sending a hidden field with the same name as the multiple select (in our case, `entry[:moods][]`) but a blank value, allows for a multiple select form to still be updated with all options deselected. 
   * I bypassed the creation of the new Mood instance with an empty name value in my code with a simple `if` statement:

```
if !mood.empty?
    @entry.moods << Mood.find_or_create_by(name: mood)
 end
```

While writing validations to ensure that a user cannot see or edit another User's data, I ran into an interesting problem. While I could control for a user manually entering a URL to view a post that does not belong to them:

```
@entry = Entry.find(params[:id])
  if current_user && @entry.user_id == current_user.id
    erb :'/entries/show'
  else
    halt erb(:error_entries)
  end
```

The code would break if a user tried to enter the ID of an entry that has been deleted from the database, or that has not yet been created, into the URL. 
![](http://drive.google.com/uc?export=view&id=19uKm7SIw_ZCKn34IIg5cMgk2AL6YHuob)

I tried writing all kinds of conditional statements to handle the non-existent Entry, such as making sure the `@entry` instance variable assignation only executes if `Entry.find(params[:id])` returns true:

```
get '/entries/:id' do
  if Entry.find(params[:id])
    @entry = Entry.find(params[:id])
     if current_user && @entry.user_id == current_user.id
       erb :'/entries/show'
     else
       halt erb(:error_entries)
     end
  end
end
```

but I kept hitting errors. With the help of a classmate, we googled the `ActiveRecord::RecordNotFound` error and ultimately found that the `#find` method always throws the `ActiveRecord::RecordNotFound` error if an object isn't found, but using `#find_by` or `#find_by_id` will return `nil` if the object isn't found. So, I resolved the error with: 

```
get '/entries/:id' do
    @entry = Entry.find_by_id(params[:id])

    if @entry.nil?
      redirect "/entries"
    elsif current_user && @entry.user_id == current_user.id
      erb :'/entries/show'
    else
      halt erb(:error_entries)
    end
  end
```

## The Work Continues
I'm excited to have turned in my first Sinatra application, and there are features that I want to continue to develop for this project. I would like to:
* Deploy my project to Heroku
* Utilize the Sinatra Flash gem to display error messages
* Dive deeper into the front end styling
* Add insights to a User's dashboard page drawing correlations between recent moods / activities over a period of time

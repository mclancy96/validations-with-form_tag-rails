# Validations with `form_tag`

Now that we've learned to handle the server side of validations, we need to
take care of the client side.

At this point, we'll be in step three of the following flow:

1. User fills out the form and hits "Submit", transmitting the form data via
   a POST request.
2. The controller sees that validations have failed, and re-renders the form.
3. **The view displays the errors to the user**.

# Objectives

After this lesson, you'll be able to...

- Prefill in form values based on an instance
- Print out full error messages based on an invalid instance
- Introspect on errors for a field
- Apply an error class to invalid fields


# Pre-Filling Form Values

No one likes re-doing work. First, let's make sure we know how to pre-fill
forms with the user's input so they don't have to type everything all over
again.

In the next lesson, `form_for` will be handling this for us, but `form_for`
is *very* heavy on Rails magic, and continues to baffle scientists to this day,
so we're going to do things by hand first, to understand what's really going on
and what to do when tools like `form_for` don't work.

Let's start with a vanilla form, using the [FormTagHelper][form_tag_helper]:

[form_tag_helper]: http://api.rubyonrails.org/classes/ActionView/Helpers/FormTagHelper.html

```erb
<!-- app/views/people/new.html.erb //-->

<%= form_tag("/people") do %>
  <div class="field">
    <%= label_tag "name", "Name" %>
    <%= text_field_tag "name" %>
  </div>
  <div class="field">
    <%= label_tag "email", "Email" %>
    <%= text_field_tag "email", @person.email %>
  </div>
  <%= submit_tag "Create Person" %>
<% end %>
```

Let's say we're working with this `Person` model:

```ruby
# app/models/person.rb

class Person < ActiveRecord::Base
  validates :name, format: { without: /[0-9]/, message: "does not allow numbers" }
  validates :email, uniqueness: true
end
```

This means validation will fail if we put numbers into the "Name" field, and
the form will be re-rendered with the invalid `@person` object available.

Now, let's plug the input back into the form:

```erb
<%= form_tag("/people") do %>
  <div class="field">
    <%= label_tag "name", "Name" %>
    <%= text_field_tag "name", @person.name %>
  </div>
  <div class="field">
    <%= label_tag "email", "Email" %>
    <%= text_field_tag "email", @person.email %>
  </div>
  <%= submit_tag "Create" %>
<% end %>
```

[text_field_tag]: http://api.rubyonrails.org/classes/ActionView/Helpers/FormTagHelper.html#method-i-text_field_tag

As you can see from the [docs][text_field_tag], the second argument to
`text_field_tag`, as with most form tag helpers, is the default value.

What's nice about this technique is that we can use the **same** form code for
empty *and* pre-filled forms, because `@person = Person.new` will create an
empty model object whose attributes are all `nil`.

# Displaying All Errors With `errors.full_messages`

The simplest way to show errors is to just spit them all out at the top of the
form by iterating over `@person.errors.full_messages`. But first, we'll have to
check whether there are errors to display with `@person.errors.any?`.

```erb
<% if @person.errors.any? %>
  <div id="error_explanation">
    <h2>There were some errors:</h2>
    <ul>
      <% @person.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

This is nice, but it's not very helpful from a user interface standpoint. It
would be much better if the incorrect fields themselves were highlighted
somehow.

# Displaying Per-Field Errors With `errors[]`

[ActiveModel::Errors][activemodel_errors] has much more than just a list of
`full_message` error strings. It can also be used to access field-specific
errors by interacting with it like a hash. If the field has errors, they will
be returned in an array of strings:

[activemodel_errors]: http://api.rubyonrails.org/classes/ActiveModel/Errors.html

```ruby
@person.errors[:name] #=> ["does not allow numbers"]
@person.errors[:email] #=> []
```

With this in mind, we can conditionally "error-ify" each field in the form,
starting with what we had before:

```erb
<div class="field">
  <%= label_tag "name", "Name" %>
  <%= text_field_tag "name", @person.name %>
</div>
```

And ending up with:

```erb
<% if @person.errors[:name].empty? %>
  <div class="field">
<% else %>
  <div class="field_with_errors">
<% end %>
  <%= label_tag "name", "Name" %>
  <%= text_field_tag "name", @person.name %>
</div>
```

# The Whole Picture

By now, our full form has grown quite a bit:

```erb
<%= form_tag("/people") do %>
  <% if @person.errors.any? %>
    <div id="error_explanation">
      <h2>There were some errors:</h2>
      <ul>
        <% @person.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <% if @person.errors[:name].empty? %>
    <div class="field">
  <% else %>
    <div class="field_with_errors">
  <% end %>
    <%= label_tag "name", "Name" %>
    <%= text_field_tag "name", @person.name %>
  </div>

  <% if @person.errors[:email].empty? %>
    <div class="field">
  <% else %>
    <div class="field_with_errors">
  <% end %>
    <%= label_tag "email", "Email" %>
    <%= text_field_tag "email", @person.email %>
  </div>

  <%= submit_tag "Create" %>
<% end %>
```

Notice that some whitespace has been added for "breathing room" and increased
readability. Additionally, indentation has been very carefully maintained.

It's already starting to feel pretty unwieldy to manually manage all of this
conditional display logic, but without an understanding of the dirty details,
we can't even begin to use more powerful tools like `form_for` correctly.

Next, we'll dive into a lab using `form_tag` and artisinally craft our own
markup.

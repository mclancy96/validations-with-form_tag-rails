# Validations with `form_with`

## Introduction

In modern Rails, the recommended way to build forms is with the `form_with` helper. This lesson will guide you through handling validations and user feedback using `form_with`, ensuring your forms are user-friendly and robust.

You'll learn how to:

- Pre-fill form values based on user input and model instances
- Display validation errors clearly to users
- Highlight invalid fields for better user experience
- Use best practices for forms in Rails 7.1

Now that we've learned to handle the server side of validations, we need to
take care of the client side.

At this point, we'll be in step three of the following flow:

1. User fills out the form and hits "Submit", transmitting the form data via
   a POST request.
2. The controller sees that validations have failed, and re-renders the form.
3. **The view displays the errors to the user**.

## Objectives

After this lesson, you'll be able to...

- Prefill in form values based on an instance
- Print out full error messages based on an invalid instance
- Introspect on errors for a field
- Apply an error class to invalid fields

## Pre-Filling Form Values

No one likes re-doing work. First, let's make sure we know how to pre-fill
forms with the user's input so they don't have to type everything all over
again.

Let's start with a vanilla form (no pre-filled values yet), using the
[`form_with` helper](https://api.rubyonrails.org/v7.1.0/classes/ActionView/Helpers/FormHelper.html#method-i-form_with):

```erb
<!-- app/views/people/new.html.erb //-->

<%= form_with url: "/people", local: true do |form| %>
  Name: <%= form.text_field :name %><br>
  Email: <%= form.text_field :email %>
  <%= form.submit "Create Person" %>
<% end %>
```

Here's what the HTML output will look like:

```html
<form action="/people" accept-charset="UTF-8" method="post">
  <input name="utf8" type="hidden" value="&#x2713;" />
  <input type="hidden" name="authenticity_token" value="TKTzvQF+atT/XHG/7h48xKVdXvILdiPj83XQhn2mWBNNhvv0Oh5YfAl2LM3DlHsQjbMOFVsYEyOwj+rPaSk3Bw==" />
  Name: <input type="text" name="name" id="name" /><br />
  Email: <input type="text" name="email" id="email" />
  <input type="submit" name="commit" value="Create Person" />
</form>
```

We're working with this `Person` model:

```ruby
# app/models/person.rb

class Person < ActiveRecord::Base
  validates :name, format: { without: /[0-9]/, message: "does not allow numbers" }
  validates :email, uniqueness: true
end
```

This means validation will fail if we put numbers into the "Name" field, and
the form will be re-rendered with the invalid `@person` object available.

Remember that our `create` action now looks like this:

```ruby
# app/controllers/people_controller.rb

  def create
    @person = Person.new(person_params)

    if @person.valid?
      @person.save
      redirect_to person_path(@person)
    else
      # re-render the :new template WITHOUT throwing away the invalid @person
      render :new
    end
  end
```

With this in mind, we can use the invalid `@person` object to "re-fill" the
usually-empty `new` form with the user's invalid entries. This way they don't
have to re-type anything.

(You wouldn't _always_ want to do this –– for example, with credit card numbers ––
because you want to minimize the amount of times sensitive information travels
back and forth over the internet.)

Now, let's plug the information back into the form using `form_with` and pre-fill the values from the `@person` instance:

```erb
<!-- app/views/people/new.html.erb //-->

<%= form_with model: @person, local: true do |form| %>
  Name: <%= form.text_field :name %><br>
  Email: <%= form.text_field :email %>
  <%= form.submit "Create Person" %>
<% end %>
```

As you can see from the [docs for `form_with`](https://api.rubyonrails.org/v7.1.0/classes/ActionView/Helpers/FormHelper.html#method-i-form_with), using the `model:` option automatically pre-fills the form fields with the values from the model instance. The HTML for the two field inputs used to look like this:

```html
Name: <input type="text" name="name" id="name" /><br>
Email: <input type="text" name="email" id="email" />
```

But now it will look like this:

```html
Name: <input type="text" name="person[name]" id="person_name" value="Jane Developer" /><br />
Email: <input type="text" name="person[email]" id="person_email" value="jane@developers.fake" />
```

When the browser renders those inputs, they'll be pre-filled with the data in
their `value` attributes.

This is the same technique used to create `edit`/`update` forms.

We can also use the **same** form code for empty _and_ pre-filled forms
because `@person = Person.new` will create an empty model object whose
attributes are all `nil`.

## Displaying All Errors With `errors.full_messages`

When a model fails validation, its `errors` attribute is filled with
information about what went wrong. Rails creates an
[ActiveModel::Errors](https://api.rubyonrails.org/v7.1.0/classes/ActiveModel/Errors.html) object to carry this information.

The simplest way to show errors is to just display them at the top of the
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

If the model has two errors, there will be two items in `full_messages`, which
could result in the following HTML:

```html
<div id="error_explanation">
  <h2>There were some errors:</h2>
  <ul>
    <li>Name does not allow numbers</li>
    <li>Email is already taken</li>
  </ul>
</ul>
```

This is nice, but it's not very helpful from a user interface standpoint. It
would be much better if the incorrect fields themselves were highlighted
somehow.

## Displaying Pre-Field Errors With `errors[]`

`ActiveModel::Errors` has much more than just a list of
`full_message` error strings. It can also be used to access field-specific
errors by interacting with it like a hash. If the field has errors, they will
be returned in an array of strings:

```ruby
@person.errors[:name] #=> ["does not allow numbers"]
@person.errors[:email] #=> []
```

With this in mind, we can conditionally "error-ify" each field in the form,
targeting the divs containing each field. With `form_with`, you can use the form builder to generate fields and labels, and add error classes as needed:

```erb
<div class="field<%= ' field_with_errors' if @person.errors[:name].any? %>">
  <%= form.label :name, "Name" %>
  <%= form.text_field :name %>
</div>
```

You can override `ActionView::Base.field_error_proc` to change it to something
that suits your UI. It's currently defined as this within `ActionView::Base:`.

**Note:** There is a deliberate space added in `' field_with_errors'` in the
example above. If `@person.errors[:name].any?` validates to true, the goal here
is to produce two class names separated by a space (`class=field
field_with_errors`). Without the added space, we would get
`class=fieldfield_with_errors` instead!

## The Whole Picture

By now, our full form has grown quite a bit:

```erb
<%= form_with model: @person, local: true do |form| %>
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

  <div class="field<%= ' field_with_errors' if @person.errors[:name].any? %>">
    <%= form.label :name, "Name" %>
    <%= form.text_field :name %>
  </div>

  <div class="field<%= ' field_with_errors' if @person.errors[:email].any? %>">
    <%= form.label :email, "Email" %>
    <%= form.text_field :email %>
  </div>

  <%= form.submit "Create" %>
<% end %>
```

Notice that some whitespace has been added for "breathing room" and increased readability. Additionally, indentation has been very carefully maintained.

As you can see, manually managing all of this conditional display logic can become unwieldy as forms grow more complex. Understanding these details is important, but Rails provides advanced helpers to make working with forms easier and less error-prone.

`form_with` is the recommended and most flexible approach for building forms. It combines the best features of previous helpers and is designed to be the standard going forward.

**Key takeaways:**

- `form_with` is the modern, recommended helper for all new Rails applications.
  - It works for both model-based and non-model forms.
  - It provides a clean, consistent API and integrates well with Rails' validation and error handling features.

In this lesson, you learned the fundamentals of form handling and validation feedback using `form_with`. This foundation will help you build robust, user-friendly forms in your Rails applications.

Next, we'll dive into a lab using `form_with` and artisanally craft our own markup, putting these concepts into practice.

# Validations with `form_tag`

## Objectives

- render or redirect based on validation of instance in create/update
- Prefill in form values based on an instance
- Print out full error messages based on an invalid instance
- introspect on errors for a field
- apply an error class to a field conditionally based on errors on a field

## Notes

Validating a form submission begins with the invalid form data being submitted to a controller action. The controller action processing the form must validate the instance through trying to save it or update it and then when invalid, the controller action must render a form, preserving the instance in memory for this request.

Once you render the form, you have to introspect on the object to figure out how it's invalid.

#full_messages
#errors
#errors[:field]

Show the lower level methods that allow granular control of error data.

demonstrate how to print out the full error messages and even customize them a bit during printing.

prefill the form field values based on the error instance. you could even show them how to not write out the invalid pre-filled data should they want, say, in the case of a credit card number.

demonstrate how to set a class "error" (we should have a stylesheet that has a .error giving form inputs a red border) if the object has an error on that particular field.

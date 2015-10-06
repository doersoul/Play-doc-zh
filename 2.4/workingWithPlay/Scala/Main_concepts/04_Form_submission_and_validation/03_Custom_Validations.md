#Using Custom Validations
The [validation package](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/package.html) allows you to create ad-hoc constraints using the `verifying` method. However, Play gives you the option of creating your own custom constraints, using the [`Constraint`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/Constraint.html) case class.

Here, we’ll implement a simple password strength constraint that uses regular expressions to check the password is not all letters or all numbers. A [`Constraint`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/Constraint.html) takes a function which returns a [`ValidationResult`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/ValidationResult.html), and we use that function to return the results of the password check:

```scala
val allNumbers = """\d*""".r
val allLetters = """[A-Za-z]*""".r

val passwordCheckConstraint: Constraint[String] = Constraint("constraints.passwordcheck")({
  plainText =>
    val errors = plainText match {
      case allNumbers() => Seq(ValidationError("Password is all numbers"))
      case allLetters() => Seq(ValidationError("Password is all letters"))
      case _ => Nil
    }
    if (errors.isEmpty) {
      Valid
    } else {
      Invalid(errors)
    }
})
```

> **Note**: This is an intentionally trivial example. Please consider using the [OWASP guide](https://www.owasp.org/index.php/Authentication_Cheat_Sheet#Implement_Proper_Password_Strength_Controls)  for proper password security.

We can then use this constraint together with [`Constraints.min`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/Constraints.html) to add additional checks on the password.

```scala
val passwordCheck: Mapping[String] = nonEmptyText(minLength = 10)
  .verifying(passwordCheckConstraint)
```
#使用自定义验证器
[validation 包](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/package.html) 允许你调用`verifying` 方法创建特殊的约束。但是, Play 还给你一个可选的创建自定义约束的方法，是使用[`Constraint`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/Constraint.html) 样例类。

这里, 我们会实现一个简单的密码强度约束，通过使用正则表达式来检查密码是不是全为字母或全为数字。一个[`Constraint`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/Constraint.html) 接受一个返回[`ValidationResult`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/ValidationResult.html)的函数为参数, 并且用这个函数来返回密码检查的结果:

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

> **注意**: 这只是一个特意演示的例子。对于正确的密码安全设计请使用[OWASP 指南](https://www.owasp.org/index.php/Authentication_Cheat_Sheet#Implement_Proper_Password_Strength_Controls) 。

我们可以使用[`Constraints.min`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/Constraints.html)结合这个约束，来给密码添加额外的验证。

```scala
val passwordCheck: Mapping[String] = nonEmptyText(minLength = 10)
  .verifying(passwordCheckConstraint)
```
Validoctor is an all-purpose data validator for backend java projects. It performs validation basing on rules passed 
along with object to be validated. It can operate depending on several traits and on various kinds of rules in an effort 
to cater to specific needs of all projects.

# Getting started
To add validoctor to your project:
```groovy
repositories {
    jcenter()
}
dependencies {
  compile 'com.miquido.validoctor:validoctor:1.0.0'
}
```
Validoctor is a data validation library handling validations on complex data structures. It is designed to be 
used as a standalone complex validation solution, and not for cooperation with Bean Validation or any other 
validation solution.

Basic concepts and a production-grade example are shown below. For more specific use cases, please refer to tests. 
When in doubt about how something should be used, answer is usually easily found there.

# Vocabulary
* Ailment - like a single violation of a Rule. It carries a meaningful name and may have additional parameters.
* Diagnosis - validation report stating whether object is valid or not and containing all Ailments discovered in it. 
It is structured to be useful for any client code that may want to read it, so it may be used directly as, for example, 
a response body.

# Traits
Validoctor instance can be created using a builder:

```java
Validoctor validoctor = Validoctor.builder().pedantic(true).exceptional(false).build();
```

For now, there are two traits.

* Pedantic Validoctor will go through all rules to the end and return a complete Diagnosis with all violations. One who is not pedantic will stop on first violation encountered and return this one only.
* Exceptional Validoctor will throw an exception containing a Diagnosis instead of returning it, if it finds any violations.

Pedantic and non-exceptional Validoctor is the default obtained by calling build() without specifying any traits.

# Rules
Each rule provides two things: definition of how validity is tested and Ailment that should be stated upon violation of the rule.
There are 3 types of rules, calling validation with each type is the same:

```java
Diagnosis diagnosis = validoctor.examine(patient, RULE_1, RULE_2);
```

examine() method accepts any number of rules of given type. There also are examineCombo methods that allow passing some permutations of rules of several kinds:

```java
Diagnosis diagnosis = validoctor.examineCombo(patient, Rules.notNull(), multiRule1, multiRule2);
```

Types are:

* Just Rule - simple interface used to define validations of one value
* MultiRule - rule sets that allow defining validations for whole data structures on per property basis
* ReducerRules - rules that are applied to a set of properties of the same type that is reduced to one value

# Example
This is a description of slightly simplified ComplexCase1Test class as it shows and tests validation of 
production-grade complexity.

Imagine you need to validate an instance of a Product defined like this:
```java
data class NutritionFacts(var kcal: Int?, 
                          var fibre: Double?, 
                          var protein: Double?, 
                          var fat: Double?)

data class Comment(var authorId: Long?, 
                   var text: String?, 
                   var upVotedUsersIds: List<Long>?, 
                   var downVotedUsersIds: List<Long>?)

data class Product(var name: String?, 
                   var skuId: String?, 
                   var description: String?, 
                   var weightG: Float?, 
                   var volumeMl: Float?,
                   var nutritionFacts: NutritionFacts?, 
                   var glutenFree: Boolean?, 
                   var vegan: Boolean?,
                   var comments: List<Comment>?, 
                   var reviewScores: MutableList<Int>?)
```

Let's start with thinking about what we want to validate. Thinking on a per-class basis is the recommended approach 
for using Validoctor. So, starting with NutritionFacts class, we surely want all of the values to be positive. 
We write a simple MultiRule for that:
```java
val nutritionFactsRules: MultiRule<NutritionFacts> =
        MultiRule.builder<NutritionFacts>().reflexiveProperties(NutritionFacts::class.java)
            .addRulesForAll(Number::class.java, numberPositive())
            .build()
```
What we did here? MultiRule is a list of rules that are applied to specified fields of the validated object. 
By using reflexiveProperties method on MultiRuleBuilder we allowed the resulting MultiRule to use reflection to find the 
values of the fields to apply the rules to. Then, we used addRulesForAll to tell the rule that we want it to read all 
fields of type Number (or its subtypes) and apply the numberPositive rule to each of them. It is a predefined rule 
available in Rules class that just checks if number is larger than 0.

Ok, so that is what we want to validate in NutritionFacts. Let's move on to Comment class. We certainly need a valid, 
non-null authorId and we want the text of the comment to be not empty, not longer that certain characters count and not 
contain any inappropriate words. For that, we will first need to define our own custom censor rule like this:
```java
val stringCensorRule: SimpleRule<String> =
        SimpleRule("CENSORED_WORD", Predicate { str -> str == null || !str.contains("fuck", true) }, Severity.WARN)
```
In case when there is no appropriate predefined rule available in Rules class, we can create our own rules as shown above. 
Using SimpleRule constructor should cover 99.99% cases, so if you find yourself wanting to do something more complicated 
it might be a sign that you are trying to over-engineer. What we did here is we specified a name of our custom rule, 
passed a predicate that will be applied to patients to determine whether they are valid or not, and specified severity 
of violation of this rule. We decided on just WARN and not ERROR as we assume Comments reported to violate this rule 
will need to be reviewed by moderators and not just plain rejected.

Now, we are ready to create a MultiRule for Comment object that will specify all validations we need:
```java
val commentRules: MultiRule<Comment> = 
        MultiRule.builder<Comment>().reflexiveProperties(Comment::class.java)
            .addRules("authorId", notNull(), numberPositive())
            .addRules("text", notNull(), stringTrimmedNotEmpty(), stringMaxLength(500), stringCensorRule)
            .build()
```
We used MultiRuleBuilder just like for NutritionFacts. This time though, we specify fields we want the rules to be 
applied to one by one, by their name. If we did not use reflexiveProperties, we would need to also specify the getters 
for these fields. Each addRules call will make resulting rule apply all the specified rules to given field. So here, 
authorId will be checked if it is not null and then if it is a positive number, and text will be checked for nullity, 
not emptiness, max allowed length and inappropriate words with our custom censor rule.

And now, for the biggest task: we need to apply a series of various validations to fields of Product class. For better 
readability, we can decide to split validations of such complex classes into a few MultiRules. Let's do that here and 
first specify rules that deal exclusively with nullity of Product's fields:
```java
val nullityRules: MultiRule<Product> = 
        MultiRule.builder<Product>().reflexiveProperties(Product::class.java)
            .addRulesForAll(Boolean::class.java, notNull())
            .addRulesForAll(Float::class.java, notNull())
            .addRules("name", notNull<String>())
            .addRules(Predicate { p -> p.skuId != null }, "nutritionFacts", notNull<NutritionFacts>())
            .addRules(Predicate { p -> p.skuId == null }, "nutritionFacts", isNull<NutritionFacts>())
            .build()
```
What we did here is we required all Booleans (glutenFree and vegan fields) and all Floats (weightG, volumeMl) to be not 
null. Then, we also specified that we need name field to not be null (notNull predefined rule needs a type parameter 
in Kotlin if there are no other rules passed that allow inferring the type of field). Last two calls to addRules deal 
with nutritionFacts fields, and differ from what we have seen so far in that they accept an additional Predicate as 
first argument. Those are conditional rules, that will only be applied if that predicate is fulfilled. This allows us 
to require the nutrition facts are present only if we also have the skuId of the product, and are null otherwise.

Now, we also need to validate a bunch of other stuff on Product object. Let's look at the last MultiRule we need:
```java
val validityRules: MultiRule<Product> = 
        MultiRule.builder<Product>().reflexiveProperties(Product::class.java)
            .addRules("name", stringTrimmedNotEmpty(), stringMaxLength(40))
            .addRules("skuId", stringExactLength(10), stringAlphanumeric())
            .addRules("description", stringMaxLength(200))
            .addRulesForAll(Float::class.java, numberPositive())
            .addMultiRule({ p -> p.nutritionFacts != null }, "nutritionFacts", nutritionFactsRules)
            .addRules("tags", collectionNotEmpty())
            .addMultiRuleForElements("comments", commentRules)
            .addRules("reviewScores", each(numberInRange(1, 5)))
            .build()
```
AddRules and addRulesForAll usages here are nothing new at this point, they just use some more predefined rules that can 
be found in Rules class. New hot stuff is addMultiRule method. It specifies that nutritionFacts field will be validated 
with the MultiRule we created for NutritionFacts objects at the beginning of this example. You can nest MultiRules for 
fields of complex types like this, down to hierarchies of unlimited depth. In this case, validation of the 
nutritionFactsRule is also conditional, only applied if the nutritionFacts field is not null. Another important case here 
is addMultiRuleForElements we used for list of Comment objects in Product. This method allows us to apply the MultiRule 
we defined above for Comment to each element of the list. The final thing to note is the last addRules call that has a 
Rule built using each() passed. Rules.each() allows us to achieve validation for individual elements of collection same 
as addMultiRuleForElements, it is just easier to use when we do not need a MultiRule for that, and just SimpleRules are 
enough - typically when collection elements are of primitive or String type.

With that we have defined all the validation we need for given data structure. To perform the validation on an actual 
object, we just need one call:
```java
val diagnosis = validoctor.examine(product, nullityRules, validityRules)
```
And that's it. Diagnosis object returned by our validoctor instance contains the result and all the Ailments found in 
the object.


# Dependencies
None.

# In next releases
* Add missing toString and equals/hashCode methods - DONE.
* Return rule parameters and actual patient value in Ailment for easier result handling by clients - DONE.
* Make Ailments found in elements of collection be mapped under jsonpath-compliant keys.
* Add trait for multitheaded validation.
* Write quick start guide. - DONE (example above).
* Create develop branch. - DONE.
* Building snapshots to bintray from master.
* Convert to Kotlin.

# Implicits Organizer

This project contains macros which can be helpful with handling implicits.

Common problem in scala is that if you are using a lot of type classes in project on huge case classes you can experience 'Method too large error'.
Typical solution for this is to manually add implicits to tell compiler to store intermediate results and not generate whole type class instance at once.
Sometimes it can be problematic because it requires to write a lot of boilerplate with cached implicits.

Solution for this is using one of these 2 annotations:

```scala
    @generateAllImplicits(typeClassNames: Seq[String], imports: Seq[String])

    @generateCachedImplicits(typeClassNames: Seq[String])
```
@generateAllImplicits is more powerful annotation. This annotation generate implicit instances for specified typeclass names and for all specified case class names:

```scala
    object ImplicitsWrapperSupport {
      @generateAllImplicits(
        Seq("EncoderTypeClass", "DecoderTypeClass"), //Names of Typeclass for which implicits should be generated
        Seq("import com.macros.CaseClassInspector", "import com.macros.ImplicitsGeneratorDomainTest._") // Additional imports required for macro to find domain classes
      )
      val allNestedClasses: List[CaseClassDescription] =
        CaseClassInspector.findAllCaseClasses[TestWrapperClass] // macro to get all nested case class names
    
      trait ImplicitsWrapper
        extends ImplicitsWrapperSupport.EncoderTypeClassImplicitsCache // these 2 traits were generated by @generateAllImplicits annotation
        with ImplicitsWrapperSupport.DecoderTypeClassImplicitsCache
    }
```
ImplicitsWrapper trait will contain all implicits for given Typeclasses and intermediate case class names.

For getting intermediate case class names is responsible CaseClassInspector macro.
This macro should return all nested case class names for given type.

Given this domain:

```scala
    case class TestWrapperClass(testClass1: TestClass1, testClass2: Option[TestClass2])
    case class TestClass1(int: Int, List[TestClass3])
    case class TestClass2(str: String)
    case class TestClass3(a: String)
```

The result of 'CaseClassInspector.findAllCaseClasses[TestWrapperClass]' is:

```scala
    List(
      CaseClassDescription("testWrapperClass", "TestWrapperClass"),
      CaseClassDescription("testClass1", "TestClass1"),
      CaseClassDescription("testClass2", "TestClass2"),
      CaseClassDescription("testClass3", "TestClass3")
    )
```
In this case trait 'ImplicitsWrapperSupport.EncoderTypeClassImplicitsCache' should generate code like this:
```scala
    trait EncoderTypeClassImplicitsCache {
      implicit lazy val testWrapperClassEncoderTypeClass: EncoderTypeClass[TestWrapperClass] = shapeless.cachedImplicit
      implicit lazy val testClass1EncoderTypeClass: EncoderTypeClass[TestClass1] = shapeless.cachedImplicit
      implicit lazy val testClass2EncoderTypeClass: EncoderTypeClass[TestClass2] = shapeless.cachedImplicit
      implicit lazy val testClass3EncoderTypeClass: EncoderTypeClass[TestClass3] = shapeless.cachedImplicit
    }
```
As you can see combination of these macros is very powerful. It produces cached implicits for all nested case classes.
We do not need to worry about 'Method too large error' anymore if we make sure that we specify classes for which are created a lot of typeclass instances

Typical common solutions for handling 'method too large error' is to implement annotation which is used on case class and this annotation create implicit in companion object ONLY for this case class. It means that you need to add this annotation manually for all nested case classes.
For projects with small domain it is acceptible but in bigger projects using such annotation would be USELESS. Why?

* You need to make sure that each nested case class is appropriately marked,
* You need to add a lot of imports into domain to generate typeclass instances - domain module will compile a LOT of time,
* Implicits in companion objects are very slow, because compilator looks first for implicits in scope. Looking for implicits in companion object is the last step. To fix this you will need to add manually a lot of imports to tell compiler there to find implicits.

To summarize @generateAllImplicits annotation:

* It is very good solution if your project has a very big domain,
* It will generate cached implicits for all nested case classes for you. You need to only specify main case classes and CaseClassInspector macro will find all nested case classes,
* It is good if you are struggling a lot with 'method too large error',
* If you are using a lot of different typeclasses then I recommend to create modules per each typeclass and use this annotation. Code will be compiled in parallel.

But if your project is not big then you can use below solution and annotation:

```scala
    @generateCachedImplicits(typeClassNames: Seq[String])
```

* This macro helps generate implicit val cached implicits for declared typeclasses,
* It can be useful with handling 'method too large error' in smaller projects,
* This error can occur with case classes implemented in project contain many fields and have many nested levels,
* To deal with this problem there should be created intermediate cached implicit values, so the compiler does not have to create whole class at one time, but can store intermediate results.

Usage:
 
This code:
```scala
    @generateCachedImplicits(Seq("Decoder"))
```
Should produce:

```scala
    case class TestClass(field: String)
    object TestClass {
      implicit lazy val testClassDecoder: Decoder[TestClass] = shapeless.cachedImplicit
    }
```
Cached implicits are stored in companion object.
As mentioned before this solution has some disadvantages. 
If the project is big and there are a lot of case classes then compilator can have problems with finding appropriate cached implicits.
Compilator first looks for implicits in scope and later it searches in companion objects.
To help compilator find appropriate implicits you will need to explicitly import cached implicits into scope.
Like in above example:

```scala
    import TestClass._
```

# Installation
```scala
    libraryDependencies ++= Seq(
      "mrlibs" %% "implicits-organizer-macros" % "1.0.1"
    )
```
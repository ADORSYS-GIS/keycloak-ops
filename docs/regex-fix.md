# Documentation: Fixing the Regex Pattern Error in `SmsService.java`

## 1. The Issue

In the `SmsService.java` file, there was a bug in the static initializer block that caused the application build to fail.

**Problematic Code:**
```java
final var allowedCountries = Pattern.compile(Utils.getEnv(ConfigKey.ALLOWED_COUNTRY_PATTERN, "*"), Pattern.CASE_INSENSITIVE).asMatchPredicate();
```

The issue is with the default value `"*"` provided to the `Pattern.compile()` method. In regular expressions, the `*` character is a quantifier that must follow another character. Using it alone is a syntax error.

## 2. The Error

This bug caused a `java.util.regex.PatternSyntaxException` with the message: `Dangling meta character '*'`. This exception was thrown when the `SmsService` class was loaded, leading to a fatal `ExceptionInInitializerError` that stopped the build process.

## 3. The Fix

The solution is to replace the invalid default pattern `"*"` with a valid one that correctly represents the intended behavior ("allow all countries by default").

**Corrected Code:**
```java
final var allowedCountries = Pattern.compile(Utils.getEnv(ConfigKey.ALLOWED_COUNTRY_PATTERN, ".*"), Pattern.CASE_INSENSITIVE).asMatchPredicate();
```

**Why `".*"` is the correct fix:**
- `.` matches any single character.
- `*` matches the preceding character zero or more times.
- Together, `.*` creates a valid pattern that matches any sequence of characters, including an empty one.

This change ensures that if the `ALLOWED_COUNTRY_PATTERN` environment variable is not set, the application defaults to allowing phone numbers from all countries without causing a compilation error.
